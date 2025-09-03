pipeline {
    agent { label 'agent1' }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '339759183788'
        ECR_REPO = 'for-java-app'
        IMAGE_TAG = 'latest'
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/firstcheck-organization/jenkins.git'
            }
        }

        stage('Build') {
            steps {
                dir('javaapp-pipeline') {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                dir('javaapp-pipeline') {
                    sh 'mvn clean test'
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                dir('javaapp-pipeline') {
                    sh '''
                        wget -q https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -O html.tpl
                        trivy fs --format template --template "@html.tpl" -o report.html .
                    '''
                }
            }
        }

        stage('Sonar Analysis') {
            steps {
                dir('javaapp-pipeline') {
                    withSonarQubeEnv('sonar') {
                        sh '''
                            mvn verify sonar:sonar \
                            -Dsonar.projectKey=java-app \
                            -Dsonar.projectName=java-app 
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('javaapp-pipeline') {
                    sh 'docker build -t javaapp .'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image javaapp'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    echo "🔐 Logging in to Amazon ECR..."
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                    echo "🏷️ Tagging Docker image..."
                    docker tag javaapp:latest $ECR_URI

                    echo "📤 Pushing image to ECR..."
                    docker push $ECR_URI
                '''
            }
        }

        stage('Deploy from ECR') {
            steps {
                sh '''
                    echo "🔐 Logging in to Amazon ECR..."
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                    echo "⬇️ Pulling image from ECR..."
                    docker pull $ECR_URI

                    echo "🚀 Running container from ECR image..."

                    # Stop and remove existing container if it exists
                    docker rm -f javaapp-container || true

                    # Run the container on port 5000
                    docker run -d \
                        --name javaapp-container \
                        -p 5000:5000 \
                        $ECR_URI

                    echo "✅ Container is running at http://localhost:5000"
                '''
            }
        }
    }
}
