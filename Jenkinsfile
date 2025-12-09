pipeline {
  agent any
  environment {
    ECR_REGISTRY         = '381491999288.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_REPO           = 'udemy/devsecops'
    IMAGE_TAG            = 'latest'
    JD_ID                = "${IMAGE_REPO}:${IMAGE_TAG}"
    JD_TAGGED_IMAGE_NAME = "${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
  }
  tools { 
        maven 'Maven-Default'  
  }
  stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=jm-devsecops_jm -Dsonar.organization=jm-devsecops -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=95979120b8a80dac31363d4c93425a0c40e05d7f'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }

	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("${IMAGE_REPO}")
                 }
               }
            }
    }

	stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://381491999288.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:aws-credentials') {
                    app.push("latest")
                    }
                }
            }
    }
	   
  }
}










