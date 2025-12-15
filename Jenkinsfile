pipeline {
  agent any

  tools {
    maven 'Maven-Default'
    git 'Default'
  }

  environment {
    // ===== AWS/ECR =====
    AWS_ACCOUNT_ID     = credentials('aws-account-id')          // secret text
    AWS_DEFAULT_REGION = 'us-east-1'
    IMAGE_REPO_NAME    = 'udemy/devsecops'
    IMAGE_TAG          = 'latest'                                // will be overridden after checkout
    REPOSITORY_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"

    // ===== SonarCloud =====
    SONAR_PROJECT_KEY  = 'jm-devsecops_jm'
    SONAR_ORGANIZATION = 'jm-devsecops'
    SONAR_HOST_URL     = 'https://sonarcloud.io'
    
    JD_ID                = "${IMAGE_REPO}:${IMAGE_TAG}"
    JD_TAGGED_IMAGE_NAME = "${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
  }

  stages {

    // Declarative automatically does: "Declarative: Checkout SCM"

    stage('Compute Image Tag') {
      steps {
        // write short git SHA to a temp file
        sh 'git rev-parse --short HEAD > .gitsha'
        script {
          env.IMAGE_TAG = "${BUILD_NUMBER}-" + readFile('.gitsha').trim()
          echo "IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Login to AWS ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region "$AWS_DEFAULT_REGION" \
          | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com"
        '''
      }
    }

    stage('CompileandRunSonarAnalysis') {
      steps {
        withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
          // Use $VAR in a single-quoted shell block to avoid Groovy interpolation warnings
          sh '''
            mvn clean verify sonar:sonar \
              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
              -Dsonar.organization=$SONAR_ORGANIZATION \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.token=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('RunSCAAnalysisUsingSnyk') {
      steps {		
		   	withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
				  sh 'mvn snyk:test -fn'
				}
			}
    }
    
    //stage('RunSCAAnalysisUsingSnyk') {
    //  steps {
    //    withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
    //      // If using the Maven plugin instead, swap to: mvn snyk:test -fn -Dsnyk.token=$SNYK_TOKEN
    //      sh '''
    //        mvn snyk:test -fn -Dsnyk.token=$SNYK_TOKEN
    //      //  snyk auth "$SNYK_TOKEN"
    //      //  snyk test --severity-threshold=high || true
    //      //  snyk monitor || true
    //      '''
    //    }
    //  }
    //}

    stage('Build Image') {
      steps {
        script {
          dockerImage = docker.build("${IMAGE_REPO_NAME}:${env.IMAGE_TAG}")
        }
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}
          docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:latest
          docker push ${REPOSITORY_URI}:${IMAGE_TAG}
          docker push ${REPOSITORY_URI}:latest
        '''
      }
    }

    stage('Update K8s Manifests (for Argo CD)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                          usernameVariable: 'GIT_USERNAME',
                                          passwordVariable: 'GIT_PASSWORD')]) {
          sh """
            set -e
            rm -rf k8s-manifests
            git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/jtmensah/devsecops-jenkins-k8s-argocd-configurations-repo.git k8s-manifests
            cd k8s-manifests
            git checkout main

            # Update the image reference in your deployment file
            sed -i "s|image: .*|image: ${REPOSITORY_URI}:${IMAGE_TAG}|g" kubernetes/asg-deployment.yaml

            git config user.email "jenkins@yourdomain.com"
            git config user.name "Jenkins CI"
            git add kubernetes/asg-deployment.yaml
            git commit -m "Update image to ${IMAGE_TAG} - Build #${BUILD_NUMBER}" || true
            git push origin main
          """
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
