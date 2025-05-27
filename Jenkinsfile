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
                    def dockerImage = docker.build("myrepo/solar-system:${env.BUILD_NUMBER}")
                    docker.withRegistry('https://registry.example.com', 'docker-registry-creds') {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker pull myrepo/solar-system:${env.BUILD_NUMBER}'
                sh 'docker run -d -p 3000:3000 -e MONGO_URI=$MONGO_URI -e MONGO_USERNAME=$MONGO_USERNAME -e MONGO_PASSWORD=$MONGO_PASSWORD myrepo/solar-system:${env.BUILD_NUMBER}'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}