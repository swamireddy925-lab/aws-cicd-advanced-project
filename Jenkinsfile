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

        stage('Deploy to Production with Rollback') {
            steps {
                echo 'Deploying application to Production'

                sh '''
                    PREVIOUS_IMAGE=$(docker inspect --format='{{.Config.Image}}' aws-cicd-container 2>/dev/null || true)

                    if [ -z "$PREVIOUS_IMAGE" ]; then
                        PREVIOUS_IMAGE="aws-cicd-app:v1"
                    fi

                    echo "Previous Production image: $PREVIOUS_IMAGE"

                    if ! docker image inspect "$PREVIOUS_IMAGE" >/dev/null 2>&1; then
                        echo "ERROR: Previous Production image not found"
                        exit 1
                    fi

                    docker rm -f aws-cicd-container || true

                    if ! docker run -d \
                        --name aws-cicd-container \
                        -p 8081:80 \
                        --restart unless-stopped \
                        aws-cicd-app:${BUILD_NUMBER}; then

                        echo "ERROR: Production deployment failed"
                        echo "Starting rollback..."

                        docker rm -f aws-cicd-container || true

                        docker run -d \
                        --name aws-cicd-container \
                        -p 8081:80 \
                        --restart unless-stopped \
                        "$PREVIOUS_IMAGE"

                        exit 1
                    fi

                    HEALTHY=false

                    for i in 1 2 3 4 5 6 7 8 9 10
                    do
                        if curl -fsS http://localhost:9999 > /dev/null
                        then
                            HEALTHY=true
                            echo "SUCCESS: Production application is healthy"
                            break
                        fi

                        echo "Waiting for Production application..."
                        sleep 3
                    done

                    if [ "$HEALTHY" != "true" ]; then
                        echo "ERROR: Production health check failed"
                        echo "Rolling back to previous image..."

                        docker rm -f aws-cicd-container || true

                        if docker run -d \
                            --name aws-cicd-container \
                            -p 8081:80 \
                            --restart unless-stopped \
                            "$PREVIOUS_IMAGE"; then

                            ROLLBACK_HEALTHY=false

                            for i in 1 2 3 4 5 6 7 8 9 10
                            do
                                if curl -fsS http://localhost:8081 > /dev/null
                                then
                                    ROLLBACK_HEALTHY=true
                                    echo "SUCCESS: Previous Production version restored"
                                    break
                                fi

                                sleep 3
                            done

                            if [ "$ROLLBACK_HEALTHY" != "true" ]; then
                                echo "CRITICAL: Rollback health check failed"
                            fi
                        else
                            echo "CRITICAL: Could not restore previous version"
                        fi

                        exit 1
                    fi

                    echo "Production deployment completed successfully"
                '''
            }
        }
    }
}
