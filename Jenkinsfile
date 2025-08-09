pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID      = '31979881-0b34-4a3e-ac11-1e4a489e11d3'
        NETLIFY_AUTH_TOKEN   = credentials('netlify-token')
        REACT_APP_VERSION    = "1.0.$BUILD_ID"
    }

    stages {

        stage('Build app') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    echo "Building with version: $REACT_APP_VERSION"
                    REACT_APP_VERSION=$REACT_APP_VERSION npm run build
                    ls -la
                '''
            }
        }

        stage('Build my-playwright image') {
            steps {
                script {
                    // Dockerfile е в src/Dockerfile
                    docker.build('my-playwright', '-f src/Dockerfile .')
                }
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
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npx serve -s build &>/dev/null &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Local E2E',
                                allowMissing: false,
                                keepAll: false,
                                alwaysLinkToLastBuild: true,
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --site=$NETLIFY_SITE_ID --json > deploy-output.json
                '''
                script {
                    def json = readJSON file: 'deploy-output.json'
                    env.CI_ENVIRONMENT_URL = json.deploy_url
                }
                sh '''
                    echo "Running STAGING E2E on: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E',
                        allowMissing: false,
                        keepAll: false,
                        alwaysLinkToLastBuild: true,
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://peaceful-daffodil-303af5.netlify.app'
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to PRODUCTION. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --site=$NETLIFY_SITE_ID --prod
                    echo "Running PROD E2E on: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E',
                        allowMissing: false,
                        keepAll: false,
                        alwaysLinkToLastBuild: true,
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
