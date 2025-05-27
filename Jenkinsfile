pipeline {
    agent any

    environment {
        MONGO_URI = credentials('mongo-uri')
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'npm run test'
            }
            post {
                always {
                    junit 'test-results.xml'
                    cobertura coberturaReportFile: 'coverage/cobertura-coverage.xml'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Docker Build and Push') {
            steps {
                script {
                    docker.withRegistry('https://registry.example.com', 'docker-registry-creds') {
                        def dockerImage = docker.build("solar-system:${env.BUILD_NUMBER}")
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'ansible-playbook deploy-playbook.yml'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}