pipeline {
  agent any
  tools {
        maven 'Maven_3_5_2'
    }
  environment {
        REPORT_EMAIL = 'olimovshahboz501@gmail.com'
        // ECR target; the push uses the EC2 instance role, so no AWS keys are stored in Jenkins
        AWS_REGION     = 'eu-north-1'
        AWS_ACCOUNT_ID = '855215820447'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO       = 'buggy-app'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {
		withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
		sh 'mvn clean verify sonar:sonar -Dsonar.token=$SONAR_TOKEN'
		}
			}
      }

    stage('RunSCAAnalysisUsingSnyk') {
            steps {
		withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
		sh 'mvn snyk:test@snyk-test'
		sh 'mvn snyk:monitor@snyk-monitor'
		}
		// Turn the JSON results into a readable HTML report
		sh 'npx --yes snyk-to-html@3.7.11 -i snyk-report.json -o snyk-report.html'
			}
      }

    stage('BuildDockerImage') {
            steps {
		sh 'docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .'
			}
      }

    stage('PushImageToECR') {
            steps {
		// Short-lived ECR login token from the instance role, piped straight into docker login
		sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
		sh 'docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG'
		sh 'docker push $ECR_REGISTRY/$ECR_REPO:latest'
			}
      }
  }
  post {
    always {
		archiveArtifacts artifacts: 'snyk-report.*', allowEmptyArchive: true
		emailext(
			to: "${REPORT_EMAIL}",
			subject: "Snyk SCA report: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${currentBuild.currentResult})",
			body: "Snyk SCA results for ${env.JOB_NAME} build #${env.BUILD_NUMBER} are attached (HTML and JSON).\nBuild: ${env.BUILD_URL}",
			attachmentsPattern: 'snyk-report.*'
		)
    }
  }
}
