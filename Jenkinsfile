pipeline {
    agent any

    environment {
        // AWS and Docker settings
        AWS_ACCOUNT_ID = credentials('aws-account-id')  // Store as Jenkins credential
        AWS_DEFAULT_REGION = 'us-east-1'
        IMAGE_REPO_NAME = 'asg'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"

        // GitHub settings for K8s manifests
        K8S_MANIFESTS_REPO = 'https://github.com/jtmensah/devsecops-jenkins-k8s-argocd-configurations-repo.git'
        K8S_MANIFESTS_BRANCH = 'main'
        K8S_MANIFESTS_PATH = 'kubernetes/asg-deployment.yaml'

        // SonarCloud settings - using credentials instead of hardcoded token
        SONAR_TOKEN = credentials('sonarcloud-token')  // This is the key change
        SONAR_PROJECT_KEY = 'jm-devsecops_jm'
        SONAR_ORGANIZATION = 'jm-devsecops'
    }

    stages {
        stage('Logging into AWS ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                }
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                script {
                    // Using SONAR_TOKEN environment variable instead of hardcoded token
                    sh """
                        sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.organization=${SONAR_ORGANIZATION} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    // Snyk already uses Jenkins credentials properly
                    withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                        sh 'snyk test --severity-threshold=high'
                        sh 'snyk monitor'
                    }
                }
            }
        }

        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Pushing to ECR') {
            steps {
                script {
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:latest"
                    sh "docker push ${REPOSITORY_URI}:${IMAGE_TAG}"
                    sh "docker push ${REPOSITORY_URI}:latest"
                }
            }
        }

        stage('Update K8s Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                                     usernameVariable: 'GIT_USERNAME',
                                                     passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            # Clone the K8s manifests repository
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${K8S_MANIFESTS_REPO#https://} k8s-manifests
                            cd k8s-manifests

                            # Checkout the appropriate branch
                            git checkout ${K8S_MANIFESTS_BRANCH}

                            # Update the image tag in the deployment file
                            sed -i "s|image: ${REPOSITORY_URI}:.*|image: ${REPOSITORY_URI}:${IMAGE_TAG}|g" ${K8S_MANIFESTS_PATH}

                            # Configure git
                            git config user.email "jenkins@yourdomain.com"
                            git config user.name "Jenkins CI"

                            # Commit and push the changes
                            git add ${K8S_MANIFESTS_PATH}
                            git commit -m "Update image to ${IMAGE_TAG} - Build #${BUILD_NUMBER}"
                            git push origin ${K8S_MANIFESTS_BRANCH}

                            # Clean up
                            cd ..
                            rm -rf k8s-manifests
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f'
        }
    }
}
