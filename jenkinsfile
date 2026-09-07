pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '531988093830'
        AWS_REGION     = 'us-east-1'
        ECR_REPO       = 'jenkins-ecr-push'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        ECR_URI        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/mohanbarani/Jenkins-ECR-push.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Authenticate to ECR') {
            steps {
                // Jenkins is not running on EC2, so there's no instance role
                // to pick up credentials from automatically. Instead, this
                // pulls an AWS access key / secret key pair from the Jenkins
                // Credentials Store.
                //
                // Set this up once in: Manage Jenkins > Credentials > System
                // > Global credentials > Add Credentials
                //   Kind: AWS Credentials
                //   ID:   aws-ecr-creds
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_URI}
                    """
                }
            }
        }

        stage('Tag Image') {
            steps {
                sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}"
                sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:latest"
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh "docker push ${ECR_URI}:${IMAGE_TAG}"
                sh "docker push ${ECR_URI}:latest"
            }
        }

        stage('Verify Image in ECR') {
            steps {
                sh "aws ecr describe-images --repository-name ${ECR_REPO} --region ${AWS_REGION} --image-ids imageTag=${IMAGE_TAG}"
            }
        }
    }

    post {
        always {
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker logout ${ECR_URI} || true"
        }
        success {
            echo "Image pushed and verified: ${ECR_URI}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed. Check the stage logs above."
        }
    }
}
