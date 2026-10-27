pipeline {
  agent any
  tools {
        maven 'Maven_3_5_2'
    }
  environment {
        REPORT_EMAIL = 'olimovshahboz501@gmail.com'
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
