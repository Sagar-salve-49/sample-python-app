pipeline {

    agent any

    environment {
        IMAGE_NAME = "sagarsalve49/sample-python-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Sagar-salve-49/sample-python-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                docker run -d --name test-container \
                -p 5000:5000 $IMAGE_NAME:$IMAGE_TAG

                sleep 10

                curl -f http://localhost:3000/health

                docker stop test-container
                docker rm test-container
                '''
            }
        }

        stage('Push Docker Image') {

            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker push $IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to EKS') {

            steps {

                sh '''
                sed -i "s|image: .*|image: $IMAGE_NAME:$IMAGE_TAG|g" \
                k8s/deployment.yaml

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Deployment') {

            steps {

                sh '''
                kubectl rollout status deployment/sample-app
                kubectl get pods
                '''
            }
        }
    }

    post {

        success {
            echo 'CI/CD Pipeline Executed Successfully'
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}
