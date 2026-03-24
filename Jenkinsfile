pipeline {
    agent any

    environment {
        // Docker image info
        DOCKER_IMAGE = "kalicharandas001/devops-gke-python-pipeline"
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Versioned tag

        // GKE cluster info
        GKE_CLUSTER = "devops-cluster"
        GKE_ZONE = "us-central1-a"
        GKE_NAMESPACE = "jenkins"

        // GCP service account to impersonate via Workload Identity
        GCP_SERVICE_ACCOUNT = "jenkins-gke-deployer@project-e0d91142-6979-4ba9-bae.iam.gserviceaccount.com"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/kalicharan-practice/devops-gke-python-pipeline.git'
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:latest -t $DOCKER_IMAGE:$IMAGE_TAG .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push $DOCKER_IMAGE:latest
                docker push $DOCKER_IMAGE:$IMAGE_TAG
                '''
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                # Authenticate to GCP using Workload Identity
                gcloud auth login --impersonate-service-account=$GCP_SERVICE_ACCOUNT

                # Get GKE cluster credentials
                gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE

                # Update deployment.yaml with the new image tag
                sed -i "s|image: $DOCKER_IMAGE:.*|image: $DOCKER_IMAGE:$IMAGE_TAG|" deployment.yaml

                # Deploy to GKE
                kubectl apply -f deployment.yaml --namespace=$GKE_NAMESPACE
                kubectl apply -f service.yaml --namespace=$GKE_NAMESPACE
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        success {
            echo "Docker images $DOCKER_IMAGE:latest and $DOCKER_IMAGE:$IMAGE_TAG pushed and deployed successfully!"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}