pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo 'Getting source code from GitHub'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image'
                sh 'docker build -t aws-cicd-app:${BUILD_NUMBER} .'
            }
        }

        stage('Verify Docker Image') {
            steps {
                echo 'Checking Docker image'
                sh 'docker images'
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying application to Staging'

                sh '''
                    docker rm -f aws-cicd-staging || true

                    docker run -d \
                    --name aws-cicd-staging \
                    -p 8082:80 \
                    --restart unless-stopped \
                    aws-cicd-app:${BUILD_NUMBER}
                '''
            }
        }

        stage('Staging Health Check') {
            steps {
                echo 'Checking Staging application health'

                sh '''
                    for i in 1 2 3 4 5 6 7 8 9 10
                    do
                        if curl -fsS http://localhost:8082 > /dev/null
                        then
                            echo "SUCCESS: Staging application is healthy"
                            exit 0
                        fi

                        echo "Waiting for application..."
                        sleep 3
                    done

                    echo "ERROR: Staging health check failed"
                    exit 1
                '''
            }
        }
    }
}
