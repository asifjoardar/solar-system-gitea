pipeline {
    agent any

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: '', description: 'Docker image tag (leave blank for auto)')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment Environment')
    }

    environment {
        NODE_ENV = "${params.DEPLOY_ENV}"
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'yourdockeruser/solar-system-gitea'
        IMAGE_TAG = "${params.DOCKER_IMAGE_TAG ?: env.BUILD_NUMBER}"
        TEST_RESULTS = 'test-results.xml'
        COVERAGE_DIR = 'coverage'
        EMAIL_RECIPIENTS = 'dev-team@example.com'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master') {
                        checkout scm
                    } else {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/${env.BRANCH_NAME}"]],
                            userRemoteConfigs: [[url: 'https://github.com/asifjoardar/solar-system-gitea.git']]
                        ])
                    }
                }
            }
        }

        stage('Install Dependencies') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm run build || echo "No build script defined, skipping..."'
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm test -- --reporter mocha-junit-reporter --reporter-options mochaFile=${TEST_RESULTS}'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: "${TEST_RESULTS}"
                    archiveArtifacts artifacts: "${TEST_RESULTS}", allowEmptyArchive: true
                }
            }
        }

        stage('Code Analysis') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm install --no-save nyc'
                sh 'npx nyc --reporter=lcov npm test || true'
                publishCoverage adapters: [coberturaAdapter(path: "${COVERAGE_DIR}/cobertura-coverage.xml")], sourceFileResolver: sourceFiles('NEVER_STORE')
            }
            post {
                always {
                    archiveArtifacts artifacts: "${COVERAGE_DIR}/**", allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                    expression { params.DEPLOY_ENV == 'production' }
                }
            }
            agent {
                docker {
                    image 'docker:24.0.2'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            environment {
                DOCKER_BUILDKIT = '1'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh '''
                        echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin $DOCKER_REGISTRY
                        docker build -t $DOCKER_REPO:$IMAGE_TAG .
                        docker push $DOCKER_REPO:$IMAGE_TAG
                        docker logout $DOCKER_REGISTRY
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                    expression { params.DEPLOY_ENV == 'production' }
                }
            }
            steps {
                echo "Deploying Docker image $DOCKER_REPO:$IMAGE_TAG to ${params.DEPLOY_ENV} environment."
                // Add your deployment logic here (e.g., kubectl, helm, docker-compose, etc.)
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build succeeded. See details at ${env.BUILD_URL}"
        }
        failure {
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build failed. See details at ${env.BUILD_URL}"
        }
        unstable {
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build is unstable. See details at ${env.BUILD_URL}"
        }
        cleanup {
            deleteDir()
        }
    }
}