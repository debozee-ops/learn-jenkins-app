pipeline {
    agent any

    environment{
        NETLIFY_SITE_ID = '96ed6a5e-80eb-4cb3-a5e2-da9f5baa6ba2'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Build') {
                agent  {
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

        stage('Tests'){
            parallel{
                stage('Unit tests'){
                        agent  {
                                docker {
                                    image 'node:18-alpine'
                                    reuseNode true
                                }
                            }

                    steps{
                        sh '''
                            echo test -f build/index.html
                            npm test
                        '''
                    }

                    post{
                            always{
                                junit 'jest-results/junit.xml'                        
                            }
                        }
                }

                stage('E2E') {
                    agent  {
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
                            npx playwright test --reporter=line
                        '''
                    }
                    
                    post{
                            always{
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                            }
                        }
                }
            }
        }

        stage('Deploy staging') {
                agent  {
                    docker {
                        image 'node:18-alpine'
                        reuseNode true
                    }
                }

                steps {
                    sh '''
                        npm install netlify-cli node-jq 
                        node_modules/.bin/netlify --version
                        echo "Deploying to staging. SITE ID: $NETLIFY_SITE_ID"
                        node_modules/.bin/netlify status
                         node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                        node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                    '''
                }
                }
        stage('Approval'){
            steps{
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }        
            }


        }

        stage('Deploy Prod') {
                agent  {
                    docker {
                        image 'node:18-alpine'
                        reuseNode true
                    }
                }

            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. SITE ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
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

            environment{
                CI_ENVIRONMENT_URL = 'https://tranquil-piroshki-85dcfb.netlify.app'
            }
            

            steps {
                sh '''
                    npx playwright test --reporter=line
                '''
            }
            
            post{
                    always{
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E', reportTitles: '', useWrapperFileDirectly: true])
                    }
            }
        }
        
    }     
}