pipeline {
    agent {
        docker {
            // Google Cloud SDK image with kubectl pre-installed
            image 'gcr.io/google.com/cloudsdktool/cloud-sdk:latest'
            args '-u root:root'
        }
    }

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

        stage('Build and Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    # Docker login inside the same credentials context
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    # Build Docker image with both latest and versioned tags
                    docker build -t $DOCKER_IMAGE:latest -t $DOCKER_IMAGE:$IMAGE_TAG .

                    # Push images
                    docker push $DOCKER_IMAGE:$IMAGE_TAG
                    docker push $DOCKER_IMAGE:latest

                    # Logout to clean credentials
                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                # Authenticate using node's Workload Identity
                gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE

                # Update deployment.yaml image tag in k8s folder
                sed -i "s|image: $DOCKER_IMAGE:.*|image: $DOCKER_IMAGE:$IMAGE_TAG|" k8s/deployment.yaml

                # Deploy all resources in k8s folder
                kubectl apply -f k8s/ --namespace=$GKE_NAMESPACE
                '''
            }
        }

    }

    post {
        always {
            echo "Pipeline finished. Clean up complete."
        }
        success {
            echo "Docker images pushed and deployed successfully: $DOCKER_IMAGE:$IMAGE_TAG"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}